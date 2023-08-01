---
title: Underhanded Cairo, pt. 1
date: 2023-08-01
author: m
tags: [cairo, security]
---

Here's a pretty simple Cairo program:

```rust
use debug::PrintTrait;

fn calc(v1: u64, v2: u64) -> (u64, u64) {
    (v1 + v2, v2 * v2)
}

fn main() {
    let mut res: u64 = 0;
    let mut counter: u64 = 1;

    loop {
        if counter > 100 {
            break;
        }

        let (res, sq) = calc(res, counter);
        counter += sq;
    };

    res.print()
}
```

What is the value of `res` that's printed at the end?

Spoiler: <abbr title="zero">Hover over these words to find out.</abbr>

The reason is that the `let res` on line 16 declares a new variable instead of reassigning to the mutable `res` from the outer scope (line 8). Once the `loop` terminates, the original `res` comes back into scope and its value of 0 is printed.

The example above is a distilled version to showcase the bug. However I've approved a pull request containing the same issue. My misunderstanding of `let`, `mut` and tuple destructuring lead me to believe it's fine. Only a diligent test caught the issue.

Stay safe.
