---
title: Mock addresses for testing
date: 2023-06-28
author: m
tags: [cairo, starknet, quick-tip]
---

When writing tests, we often need mock addresses to represent some entity like an account without actually deploying an initializing an account contract. The obvious thing to do is to use values like 1, 2, 3,... but this can lead to issues. These "addresses" are used by the test runner. In a test environment, the first contract deployed using `deploy_syscall` has the address of 1, second deployed contract has the address 2 and so on. This clash could lead to nasty hidden issues especially when testing access control.

Luckily, there's a better way how to generate mock addresses. We can take advantage of the fact that both - an address and a short string - are felts! Let's declare a bunch of descriptive utility functions that we reuse throughout the test suite:

```rust
// common.cairo

use starknet::{ContractAddress, contract_address_try_from_felt252};
use option::OptionTrait;

// mock addresses

fn admin() -> ContractAddress {
    contract_address_try_from_felt252('admin').unwrap()
}

fn badguy() -> ContractAddress {
    contract_address_try_from_felt252('bad guy').unwrap()
}

fn user_no_funds() -> ContractAddress {
    contract_address_try_from_felt252('user no funds').unwrap()
}

fn user_whale() -> ContractAddress {
    contract_address_try_from_felt252('user whale').unwrap()
}
```

We can then use `common::badguy()` in the actual tests and be sure the address value does not conflict with a deployed contract. The tests become more readable as well thanks to the descriptive names.
