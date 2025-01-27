---
title: Storing user defined types, part 1
date: 2023-04-03
author: m
tags: [starknet, cairo]
---

*Deprecation notice: Since this post was published, Starknet and Cairo have evolved. As such, parts of the post might be obsolete. For the best, most up-to-date resources, please consult the [official Starknet documentation](https://docs.starknet.io/) and the [Cairo book](https://book.cairo-lang.org/).*

When we want to store a custom `struct` in a Starknet contract, we immediatelly run into a problem. Take this simple contract for example:

```rust
#[derive(Drop, Serde)]
struct Pos2D {
    x: u16,
    y: u16
}

#[contract]
mod Where {
    use super::Pos2D;

    struct Storage {
        origin: Pos2D
    }

    #[view]
    fn get_origin() -> Pos2D {
        origin::read()
    }
}
```
{: .nolineno }

Compiling it yields an error:

```sh
error: Trait has no implementation in context: core::starknet::storage_access::StorageAccess::<pos2d::pos2d::Pos2D>
 --> contract:29:47
            starknet::StorageAccess::<Pos2D>::read(
                                              ^**^

error: Trait has no implementation in context: core::starknet::storage_access::StorageAccess::<pos2d::pos2d::Pos2D>
 --> contract:37:47
            starknet::StorageAccess::<Pos2D>::write(
                                              ^***^

Error: Compilation failed.
```
{: .nolineno }

The compiler is telling us it does not know how to *read* from and *write* to Starknet storage. This behavior is defined by the `StorageAccess` trait. It's a simple trait, containing only two functions,a `read` and `write`:

```rust
trait StorageAccess<T> {
    fn read(address_domain: u32, base: StorageBaseAddress) -> SyscallResult<T>;
    fn write(address_domain: u32, base: StorageBaseAddress, value: T) -> SyscallResult<()>;
}
```
{: .nolineno }

 It's very likely implementing `StorageAccess` for custom types won't be necessary in future versions of Cairo. The compiler will just generate the appropriate code for storage by itself. However for now, we have to take care of it manually.

Let's tackle `write` first:

```rust
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
```
{: .nolineno }

Since `Pos2D` consists of only two primitive types, all we need to do is call `storage_write_syscall` two times, once for each value. The interesting part is specifying the address where to write the value to, passed in as the second parameter to the syscall. It is calculated using `storage_address_from_base_and_offset`. Notice we increment the offset by one for each struct member - it's `0_u8` for `x` and `1_u8` for `y`. We'll dive into more details of storage addressing in the second part of this series.

Now let's have a look at `read`:

```rust
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
```
{: .nolineno }

This time we use `storage_read_syscall` to retrieve the values from storage. Again, we have to pass in the correct address where from to read the values from. As before, we obtain address value using `storage_address_from_base_and_offset`. When implementing `StorageAccess`, take care that the offsets you use in `read` and `write` for each member are the same, otherwise the read and written values won't match. Final step is to wrap the structure in a `Result::Ok` and we're done.

These couple of lines are all we need to do to use `Pos2D` in Storage. It's worth mentioning we could do much more in these two functions - one obvious thing that comes to mind is value packing - two 16 bit values fit neatly into a `felt252`.

Here's an example of a contract showing the whole code, together with a simple test:

(Note: this code won't compile with the 1.0.0-alpha.6 release. Use the commit [`bc1ebcb8`](https://github.com/starkware-libs/cairo/commit/bc1ebcb8236603baa2b1f25bb04148a00aac0984) that introduced Serde derive or any other later commit to compile.)

```rust
use starknet::StorageAccess;
use starknet::StorageBaseAddress;
use starknet::SyscallResult;
use starknet::storage_read_syscall;
use starknet::storage_write_syscall;
use starknet::storage_address_from_base_and_offset;
use traits::Into;
use traits::TryInto;
use option::OptionTrait;

#[derive(Drop, Serde)]
struct Pos2D {
    x: u16,
    y: u16
}

impl Pos2DStorageAccess of StorageAccess::<Pos2D> {
    fn read(address_domain: u32, base: StorageBaseAddress) -> SyscallResult::<Pos2D> {
        Result::Ok(
            Pos2D {
                x: storage_read_syscall(
                    address_domain, storage_address_from_base_and_offset(base, 0_u8)
                )?.try_into().unwrap(),
                y: storage_read_syscall(
                    address_domain, storage_address_from_base_and_offset(base, 1_u8)
                )?.try_into().unwrap(),
            }
        )
    }

    fn write(address_domain: u32, base: StorageBaseAddress, value: Pos2D) -> SyscallResult::<()> {
        storage_write_syscall(
            address_domain, storage_address_from_base_and_offset(base, 0_u8), value.x.into()
        )?;
        storage_write_syscall(
            address_domain, storage_address_from_base_and_offset(base, 1_u8), value.y.into()
        )
    }
}

#[contract]
mod Where {
    use super::Pos2D;

    struct Storage {
        origin: Pos2D
    }

    #[view]
    fn get_origin() -> Pos2D {
        origin::read()
    }

    #[external]
    fn set_origin(x: u16, y: u16) {
        origin::write(Pos2D { x, y })
    }
}

#[test]
#[available_gas(1000000)]
fn test_pos2d_storage() {
    let origin = Where::get_origin();
    assert(origin.x == 0_u16, 'init x');
    assert(origin.y == 0_u16, 'init y');

    let (x, y) = (30_u16, 99_u16);
    Where::set_origin(x, y);

    let origin = Where::get_origin();
    assert(origin.x == x, 'set x');
    assert(origin.y == y, 'set y');
}
```
{: .nolineno }

In the second part of the series, we'll look in how to deal with embedded structs and dive more into contract storage. [Stay tuned](https://twitter.com/cairopractice).
