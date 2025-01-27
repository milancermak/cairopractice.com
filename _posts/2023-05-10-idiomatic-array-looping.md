---
title: Idiomatic array looping
date: 2023-05-10
author: m
tags: [cairo]
---

*Deprecation notice: Since this post was published, Starknet and Cairo have evolved. As such, parts of the post might be obsolete. For the best, most up-to-date resources, please consult the [official Starknet documentation](https://docs.starknet.io/) and the [Cairo book](https://book.cairo-lang.org/).*

Loops. The feature that made Cairopractors jealous of every other programming language including Brainfuck. Fortunatelly Cairo 1 fixes that. We have been promised proper for-in loops, but that's not shipped yet. For now, we have the `loop` loop.

We use loops when iterating over arrays. However to do so properly, we need to adhere to certain rules. Shout out to [Shahar for highlighting these](https://twitter.com/PapiniShahar/status/1650814998480838656).

Let's see how to do it on the Hello world of array looping:

```rust
fn arr_sum(mut vals: Span<u32>) -> u32 {
    let mut sum = 0_u32;

    loop {
        match vals.pop_front() {
            Option::Some(v) => {
                sum += *v;
            },
            Option::None(_) => {
                break sum;
            }
        };
    }
}
```

Breaking it down:

* we pass in a Span because we don't need to modify the array itself. We also mark the variable as `mut` (necessary for `pop_front`)
* on line 5, we use [`match`](https://cairo-book.github.io/ch05-02-the-match-control-flow-construct.html#matching-with-options) to properly deal with the `Option<@T>` returned by `pop_front`. BTW if you want to iterate in reverse, you can use `pop_back`
* when summing on line 7, we have to desnap the value of `v` since it's a snapshot
* the Option::None(_) block (line 9) triggers when there's no more values in the Span, i.e. the looping is done, in which case we `break` with the result `sum`
* since we don't have a `;` on line 13, the return value of the whole `loop` is the return value of the function

Notice there's no need for a index counter variable from 0 to `span.len()`. Neither do we need to get the value at the current index (`let v = vals[index]`) which saves us the use of a RangeCheck built-in on every loop iteration.

Happy looping 🔁
