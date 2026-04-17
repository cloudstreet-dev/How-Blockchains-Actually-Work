# Smart Contracts: Consensus as a Computer

Up to this point, the blockchain we've been describing has a single job: agree on an ordered list of transactions. Each transaction is a small, fixed-shape instruction — "move balance from A to B." The consensus mechanism orders them; nodes apply them to state; the state is a straightforward `dict` of balances.

Ethereum's contribution, proposed in 2013 by Vitalik Buterin and launched in 2015, was to generalize the transaction format. Instead of "move balance from A to B," a transaction could contain arbitrary code. The network would run the code as part of processing the transaction, and the code's effect on state — reading and writing arbitrary storage — would be part of the agreed-upon history.

That's it. That's the whole idea. You can describe a smart contract as "a persistent program, running on a deterministic shared virtual machine, whose state updates are committed by consensus." Or you can describe it more casually as "the chain is now a computer that everyone runs and agrees about."

## Why this is a meaningful step up

A plain payment ledger answers one question: *who owns what*. A smart-contract chain can answer *any question expressible as a program over shared state*:

- An auction: "If a higher bid arrives before block N, replace the previous bid and refund it; at block N, send the asset to the highest bidder."
- An escrow: "Hold these funds until both parties sign, or until some deadline passes."
- A multi-signature wallet: "Release funds only when 3 out of 5 designated keys have signed."
- A prediction market, a lending pool, a stablecoin, a DAO, whatever.

The key property is that the program is **deterministic** and **publicly executed**. Anyone can read the code, anyone can verify that past executions were correct, and no single party can deviate from the code — because if they did, their version of the state would disagree with everyone else's, and consensus would reject it.

This unlocks a category of application you genuinely couldn't build before: ones where the *rules themselves* are credibly neutral, because they're a function of code that everyone can see and verify, rather than a function of who happens to be running the server today.

## What a smart contract actually looks like

Here's a minimal Ethereum contract in Solidity:

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.20;

contract Counter {
    uint256 public count;

    function increment() external {
        count += 1;
    }
}
```

When you deploy this, a new contract is created at a deterministic address. That address has its own storage slot for `count`, which starts at 0. Anyone can call `increment()` by sending a transaction; each call costs gas (see below), runs the code on every full node, and increments the stored value. The `public` keyword on `count` auto-generates a view function that returns its current value, which anyone can call for free because it doesn't change state.

That's the smallest possible contract. The shape of everything bigger is the same: storage variables (a declared state layout), functions (that read/write storage), and events (logged to a side channel for off-chain indexers). Solidity is the dominant language on the EVM; Vyper is a more minimal alternative with similar semantics; Rust or Move are used on non-EVM chains. The rest of this chapter is EVM-specific because that's where the weight of the ecosystem lives, but the ideas apply broadly.

## Gas: the denial-of-service defense

If the chain is a shared computer, why can't I deploy a contract that just says `while (true) {}` and bring the whole network to a halt?

Because every operation costs **gas**, and every transaction carries a gas limit. Gas is a synthetic unit of "computational work" assigned per EVM opcode: an `ADD` costs 3 gas, a `SHA3` costs 30 gas plus 6 gas per word, an `SSTORE` (writing to persistent storage) costs thousands of gas. The transaction's sender pre-pays `gas_limit × gas_price` up front; the execution runs until it either finishes (refund the unused gas) or runs out of gas (revert all changes, but the sender still pays for the gas that was consumed).

An infinite loop therefore runs until it consumes the gas limit, at which point execution halts and the transaction reverts. The sender loses the gas they paid for, which they can't afford to do indefinitely. Gas turns "hostile program" into "self-limiting program."

Gas has two simultaneous jobs:

1. **DOS defense**, as above. Expensive operations cost proportionally more.
2. **Resource pricing / prioritization.** Gas prices are set by the market: when blocks are full, users compete to include transactions by paying higher per-gas prices. This is the fee auction that ranks transactions.

The gas price mechanism changed in Ethereum's EIP-1559 (August 2021): instead of a pure first-price auction, there's a protocol-determined `base_fee` that adjusts based on recent block fullness, plus an optional `priority_fee` (tip) to the proposer. The base fee is *burned* — permanently removed from circulation — which was a deliberate choice to decouple validator incentives from congestion.

## Why smart contracts are hard to write correctly

Ordinary programs have bugs. Smart-contract bugs have a unique combination of properties that makes them exceptionally bad:

- **Public.** The source code (or at least the bytecode, which is reverse-engineerable) is visible to attackers before it's visible to you in the wild, because attackers watch the deployment mempool.
- **Financial.** The bug is usually directly over money. There's no grace period while you patch.
- **Immutable.** Once deployed, you can't just push a fix unless you built upgradability in. Even with upgradability, there's typically a delay/governance process, during which the bug is exploitable.
- **Adversarial.** Attackers have infinite time to find the bug and zero coordination cost. If there's any way to exploit the contract, someone will find it, and they'll do it before you notice.
- **Composable.** Contracts call other contracts. A bug in a popular library reverberates through every contract using it. A secure contract can become insecure when paired with an insecure one.

History has receipts. Three worth studying:

### The DAO (June 2016) — re-entrancy

The DAO was a decentralized venture fund on Ethereum that held ~11% of all ETH at the time (about $150M). Its `withdraw()` function had the shape:

```solidity
function withdraw(uint amount) {
    if (balances[msg.sender] >= amount) {
        msg.sender.call.value(amount)();   // send ETH
        balances[msg.sender] -= amount;    // then update balance
    }
}
```

The bug is in the order. `msg.sender.call.value(amount)()` doesn't just send ETH; it triggers the receiving contract's code (its fallback function) *inside* the current execution context. If the receiver is itself a malicious contract, it can call `withdraw()` *again* before the first call's `balances[msg.sender] -= amount` line has executed. The balance check still passes — the balance hasn't been decremented yet — and the attacker withdraws again. And again. And again, until the stack limit or gas runs out.

This is **re-entrancy**, and it became the canonical Ethereum bug. The fix is the "checks-effects-interactions" pattern: update state *before* making external calls. Or, for extra paranoia, use a reentrancy guard (a mutex). The DAO bug was exploited for ~$50M, which led to the (controversial) Ethereum hard fork that reversed the theft and created Ethereum Classic.

### Parity multisig (July 2017 and November 2017) — library hygiene

The Parity multisig wallet used a shared library contract to save gas: instead of each wallet having its own copy of the code, they delegated calls to a shared library. The library had two relevant gaps:

1. An `initWallet` function that set the wallet's owners. It was supposed to be called only once during deployment, but it wasn't protected with an "only-once" modifier. An attacker in July 2017 called it on wallets that had been deployed without calling it themselves, becoming the owner and stealing ~$30M.

2. The *library itself* had an exposed `kill()` function. After the first incident, a user (accidentally!) called `initWallet` on the library contract — making themselves the owner of the library, which had never been initialized. They then called `kill()`, which ran the library's self-destruct and bricked every wallet that depended on it. About $150M of ETH became permanently inaccessible. Not stolen — just gone.

Lessons: don't ship library code with initialization functions unprotected, don't embed self-destruct in shared libraries, test what happens in every ownership state including "no owner set."

### Ronin bridge (March 2022) — key management

Not a code bug, but worth mentioning because the pattern is common and the loss was huge (~$620M). The Ronin sidechain (running the Axie Infinity game) had nine validators, of which five were required to sign cross-chain withdrawals. Four of the keys were held by Sky Mavis (the operator) and one by Axie DAO. In an earlier emergency, Sky Mavis had been granted permission to sign on behalf of the DAO as well. That delegation was never revoked.

An attacker who compromised Sky Mavis's infrastructure thus effectively had five keys and signed fraudulent withdrawals. The bug wasn't in any smart contract. It was in a human-process loop that treated "temporary delegation" as permanent.

Bridges are, in early 2026, collectively the worst-audited and most-exploited layer of the ecosystem. When you evaluate a smart-contract platform, evaluate its bridge security separately — it's a largely independent problem.

## What smart contracts are and aren't

What they are:

- **Publicly-verifiable programs** running on shared state, with rules enforced by consensus rather than by a trusted server.
- **Immutable by default** (upgradability is something you add, not something you get for free).
- **A security model** where the attacker has read access to the code, infinite time, and direct financial motivation.

What they aren't:

- A faster, cheaper general-purpose execution environment. They are the *opposite* of that: every full node re-executes every contract, so a smart-contract chain's throughput is bottlenecked on the slowest validator.
- "Smart" in any cognitive sense. They execute exactly what you wrote, bugs and all.
- Inherently more trustworthy than ordinary code. The code can be excellent, or it can be hot garbage; the consensus layer doesn't know the difference.
- Self-enforcing in the real world. A contract can move tokens on-chain; it cannot make someone deliver a package, honor a real-world contract, or update a non-blockchain database. Anything that requires a bridge to the world outside the chain needs an **oracle** — a trusted (or trust-minimized) reporter — and oracles are themselves a whole subfield with their own failure modes.

## Why throughput is limited

Because every full node re-executes every contract in every block, the chain's throughput is bounded by the resources of a single validator. You can't parallelize by adding more validators; they all do the same work. Ethereum mainnet, as of early 2026, processes roughly 15–25 transactions per second — not because the network is slow, but because that's what fits in a block that every validator can process in about 12 seconds without risking missed slots.

This is the bottleneck that led to the scaling designs we take up next. The answer isn't "make the base chain fast"; the answer is "do most of the work *off* the base chain, while inheriting its security." That's the L2 / rollup story, and it's the next chapter.
