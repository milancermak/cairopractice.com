---
title: Updating structs
date: 2023-06-02
author: m
tags: [cairo]
---

*Deprecation notice: Since this post was published, Starknet and Cairo have evolved. As such, parts of the post might be obsolete. For the best, most up-to-date resources, please consult the [official Starknet documentation](https://docs.starknet.io/) and the [Cairo book](https://book.cairo-lang.org/).*

Do you remember how, in Cairo 0, there was no concise way how to update a struct? We had to write out *all* its members even if only one changed. It was pretty annoying, as RoboTeddy pointed out in his [now legendary post](https://hackmd.io/@RoboTeddy/BJZFu56wF#Concisely-update-large-structs).

Cairo 1 fixes that. By declaring a struct as *mutable* it becomes possible to assign new values to any member of said struct:

```rust
struct Point {
    x: u64,
    y: u64,
    z: u64
}

let mut place = Point { x: 0, y: 42, z: 69 };

place.x = 33; // Point { x: 33, y: 42, z: 69 }
```

Easy and kinda obvious, isn't it? h/t to [bllu](https://twitter.com/bllu404) for teaching me this.
