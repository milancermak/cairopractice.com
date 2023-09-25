---
title: Underhanded Cairo, pt. 2. Solution
date: 2023-09-25
author: m
tags: [cairo, starknet, security]
---

The results are in. Congratulations to [wong_ssh](https://twitter.com/wong_ssh) and [0xkaliber](https://twitter.com/kaliberpoziomka), the only two participants who found the attack. Job well done guys! Thank you as well to all the others who gave it a try 🙏 So what's going on then? Where's the vulnerability?

It's a supply chain attack. There is a backdoor in the project's dependency 💥

Let's dive into the details.

The project declares a single dependency in its Scarb.toml, the [`auth` lib](https://github.com/milancermak/auth). Notice that it's using the `v0.1.0` version, but the latest version in the repo is tagged `v0.2.0`. I've done that to throw any negligent auditor off track. Even if you would inspect `auth` but forgot to checkout the commit pointed to by the `v0.1.0` tag, you'd never find the vulnerability.

The backdoor was introduced in [this `v0.1.0`](https://github.com/milancermak/auth/commit/dacdd672173087f891603a7171037387c29a4efa) commit. The `assert_owner` check will always pass when called from a contract with the address `0x61747461636b6572` (i.e. felt-encoded string "attacker"). Here's a test case to prove it:

```rust
#[test]
#[available_gas(100000000)]
fn test_hack() {
    let token: IERC20LinearDispatcher = deploy_token();
    let attacker = as_addr('attacker');
    set_contract_address(attacker);
    assert(token.balance_of(attacker).amount == 0, 'pre hack bal');
    let mint_amount: u128 = 1000000000000000;
    token.mint_to(attacker, mint_amount.into());
    assert(token.balance_of(attacker).amount == mint_amount, 'post hack bal');
}
```
{: .nolineno }

Obviously, the test shouldn't pass because `attacker` is not the owner. Yet it does, thanks to the backdoor. Simple as that.

As mentioned, this is just a proof of concept to highlight the threat of supply chain attacks. Ultimately, as builders, we are responsible for any code we ship into production. We should be deeply familiar with the libraries we use in our code.

Currently, the situation in the Cairo ecosystem is even more dangerous because Scarb doesn't yet support a lockfile. In theory, if an attacker gains access to a repo of a popular library, they could introduce a backdoor, simply change git tags and just wait until contracts with this infected library are deployed on mainnet. Thankfully, the team behind Scarb [is already working](https://github.com/software-mansion/scarb/issues/126) on lockfile support ♥️

In the meantime, as developers, we can either not use any dependencies or use tested and trusted libraries like the one from OpenZeppelin. As auditors, always remember to check the dependencies as well - hopefully they are in scope 🤞

Together, we can #keepStarknetSafe.
