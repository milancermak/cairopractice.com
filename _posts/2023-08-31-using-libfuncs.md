---
title: Using libfuncs
date: 2023-08-31
author: m
tags: [cairo]
---

You know how felt division is "disabled" by default in Cairo 1? But what if we **really** know what we're doing and are not afraid of shooting ourselves in the foot? It's actually quite easy to bring it back:

 ```rust
impl Felt252Div of Div<felt252> {
    fn div(lhs: felt252, rhs: felt252) -> felt252 {
        felt252_div(lhs, rhs.try_into().expect('div by zero'))
    }
}

use debug::PrintTrait;

fn main() {
    let r = 2 / 3;
    r.print();
    // 0x555555555555560aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaac, obviously
}
```
{: .nolineno }

The `Div` impl is using `felt252_div`, a Sierra libfunc. Since it's [already declared in corelib](https://github.com/starkware-libs/cairo/blob/0562173e14a62159f08440bd79af8ec7750fa5e1/corelib/src/lib.cairo#L169) it's straightforward to use - we don't even have to import it anymore.

By default, we can use any of the [audited](https://github.com/starkware-libs/cairo/blob/main/crates/cairo-lang-starknet/src/allowed_libfuncs_lists/audited.json) libfuncs and if we [tweak Scarb.toml](https://docs.swmansion.com/scarb/docs/extensions/starknet/contract-target.html#allowed-libfuncs-validation) we can use even the experimental ones, but only locally.

However the problem then is to 1) knowing what the libfuncs are doing and 2) how to declare and use them. For old time's sake, let's take an example of the first libfunc in the audited list: `alloc_local`. Here's a Cairo program using it:

```rust
extern fn alloc_local<u128>() -> u128 nopanic;

fn main() {
    let l: u128 = alloc_local();
}
```
{: .nolineno }


It compiles down to the following Sierra code:

```text
type [1] = u128 [storable: true, drop: true, dup: true, zero_sized: false];
type [2] = Uninitialized<[1]> [storable: false, drop: true, dup: false, zero_sized: false];
type [0] = Struct<ut@Tuple> [storable: true, drop: true, dup: true, zero_sized: true];

libfunc [1] = alloc_local<[1]>;
libfunc [3] = drop<[1]>;
libfunc [0] = struct_construct<[0]>;
libfunc [4] = store_temp<[0]>;

[1]() -> ([0]); // 0
[3]([0]) -> (); // 1
[0]() -> ([1]); // 2
[4]([1]) -> ([2]); // 3
return([2]); // 4

[0]@0() -> ([0]);
```
{: .nolineno }

But trying to `cairo-run` it fails:

```sh
Error: Failed setting up runner.

Caused by:
    #1: One of the arguments does not match the expected type of the libfunc or return statement.
```
{: .nolineno }

Why? I don't know 😅 I guess you'll have to dive into the depths of the compiler to figure that out. If you do, please [reach out](https://twitter.com/cairopractice).
