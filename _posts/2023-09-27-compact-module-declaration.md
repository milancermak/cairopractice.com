---
title: Compact module declaration
date: 2023-09-27
author: m
tags: [cairo]
---

Here's a cool technique how to declare all of a project's modules inside the project root (typically `src/lib.cairo`) without any intermediary module files.

Let's say we have a project where `src/` looks something like this:

```
.
├── core
│   ├── m1.cairo
│   ├── m2.cario
│   └── m3.cairo
├── lib.cairo
├── tests
│   ├── core
│   │   ├── test_m1.cairo
│   │   ├── test_m2.cairo
│   │   └── test_m3.cairo
│   └── utils
└── utils
    ├── common.cairo
    └── conv.cairo
```

Instead of having additional files like `core.cairo` and `utils.cairo` on the same level as `lib.cairo` that just declare the inner modules, we can put everything in `lib.cairo`:

```rust
mod core {
    mod m1;
    mod m2;
    mod m3;
}

mod utils {
    mod commmon;
    mod conv;
}

#[cfg(test)]
mod tests {
    mod core {
        mod test_m1;
        mod test_m2;
        mod test_m3;
    }
}
```
{: .nolineno }

They're the same picture!

To get a better sense of the improvement, let's have a look at some real world examples. Compare the original, verbose style that's used in [Pragma](https://github.com/Astraly-Labs/pragma-oracle/blob/fe22f63188416b5f9810f8c6b93f095a20e31c9a/src/lib.cairo) with the compact one used in [Satoru](https://github.com/keep-starknet-strange/satoru/blob/fad9f1d6b7f5d8fcf1093b873f89d2241983424e/src/lib.cairo). Notice the additional files in Pragma's `src` dir like `admin.cairo` that are there only to declare `mod admin`. I definitely like the compact one better.

Shout out to [gaetbout](https://twitter.com/gaetbout/) for showing me the way.
