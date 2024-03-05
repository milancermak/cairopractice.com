---
title: Starknet fee reduction
date: 2024-03-05
author: m
tags: [starknet]
---

Multiple factors go into calculating [transaction fees on Starknet](https://docs.starknet.io/documentation/architecture_and_concepts/Network_Architecture/fee-mechanism/). The biggest one - use of storage - is going to be addressed by 4844 in a couple of days and by [additional measures](https://community.starknet.io/t/starknet-2024-roadmap-plan-of-intent/113006) later this year.

The second major factor is the number of Cairo steps and builtins used. The heavier the computation a contract does, the more steps and builtins it consumes. A simple ERC20 `transfer` uses 1613 steps, 4 Pedersen and 47 range check builtins; an `attack` action in [Loot survivor](https://survivor.realms.world/) is roughly 82K steps and almost 6900 range check builtins. As we can see, in complex, computationaly intensive operations these calculations become a significant part of the whole transaction cost.

It's a good thing then that the gas cost per step and builtin has been steadily decreasing over the past 2 years.

First, let's compare the gas per step evolution:

| Starknet version | gas / step |
|------------------|------------|
| 0.8.0            | 0.05       |
| 0.11.0           | 0.01       |
| 0.13.0           | 0.005      |
| 0.13.1           | 0.0025     |

Version 0.8.0 was released on <abbr title="Pi day">Mar 14</abbr>, 2022. Version 0.13.1 is scheduled to hit mainnet almost exactly 2 years later, Mar 12, 2024. During this period, the StarkWare team has managed to reduce the gas per step 20 times. That's huge!

At the same time, the cost of builtins has also been going down, as denoted by the following table:

| Starknet version | bitwise | ECDSA | EC_OP | keccak | Pedersen | Poseidon | range check |
|------------------|---------|-------|-------|--------|----------|----------|-------------|
| 0.8.0            |    12.8 |  25.6 |       |        |      0.4 |          |         0.4 |
| 0.10.2           |     3.2 | 102.4 |  51.2 |        |      1.6 |          |         0.8 |
| 0.11.0           |    0.64 | 20.48 | 10.24 |  20.48 |     0.32 |     0.32 |        0.16 |
| 0.13.0           |    0.32 | 10.24 |  5.12 |  10.24 |     0.16 |     0.16 |        0.08 |
| 0.13.1           |    0.16 |  5.12 |  2.56 |   5.12 |     0.08 |     0.08 |        0.04 |

The cost of the range check builtin, which is used pretty much every time a numerical operation is done on Starknet, has decreased 10x in these past 2 years.

As an example, the aforementioned `attack` function got ~14x cheaper thanks to these reductions. Curiously, Loot Survivor is a single storage var game, so there's no other way to make the game cheaper besides paying less for computation.

There's better news still. With stuff like [dynamic layouts](https://starkware.medium.com/builtins-and-dynamic-layouts-e419a73e29e) and further improvements of the VM architecture, we can expect even more fee reduction in the future. Don't be afraid to build computationaly complex functions on Starknet, they are only going to get cheaper.
