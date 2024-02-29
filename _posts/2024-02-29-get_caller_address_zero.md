---
title: When is get_caller_address zero?
date: 2024-02-29
author: m
tags: [starknet]
---

It's very rare for the caller address (as obtained by `get_caller_address()`) to be zero on Starknet. Let's examine when can this happen.

The obvious case is when calling a view function. The call is not initiated by an account, so the caller is zero. But what about state changing functions?

In this case, the caller address is zero when it's the StarknetOS executing the function. To the best of my knowledge, this happens only in the following scenarios: invoking `__validate__` and `__execute__` in an Account contract and [handling an L1 -> L2 message](https://book.cairo-lang.org/ch15-04-L1-L2-messaging.html?#sending-messages-from-ethereum-to-starknet) in an `#[l1_handler]` function. Note that these functions are public, so _anyone_ can invoke them, but when `get_caller_address()` is zero, you can be sure it's a legit execution initiated by the protocol.

Thanks to [@gaetbout](https://twitter.com/gaetbout/), [@martiray](https://twitter.com/martriay) and [@_tserg](https://twitter.com/_tserg) for their input when researching this piece.
