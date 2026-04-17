# Proof of Work: How It Actually Works

In Chapter 4 we wrote `while not block.meets_difficulty(): block.nonce += 1` and mined a block. That loop is Proof of Work. This chapter takes it seriously: what problem is actually being solved by making that loop expensive, and why the expense is not a bug but the entire point.

## The puzzle, stated carefully

The puzzle is: given a block header `H` and a target threshold `T`, find a value `nonce` such that

<div class="math">

`sha256(H || nonce) < T`

</div>

The only input the miner varies is `nonce` (and, in practice, a few other nonce-ish fields like the coinbase transaction's arbitrary data — the extra nonce — when the primary nonce space runs out). Because `sha256` is cryptographically collision-resistant and exhibits the avalanche property, the output of each trial is effectively a uniformly random 256-bit integer. The only way to find a valid nonce is to try them, one after another, until you get lucky.

This is what Proof of Work is. It's not an algorithm in the usual sense — there's no clever approach that does better than brute force. The brute force *is* the algorithm, by design.

**Why a hash puzzle and not something else?** Because hashing has a specific combination of properties we need:

- **Unpredictable output.** You can't aim a hash. Changing the nonce by 1 changes the output in a way that is, for all engineering purposes, random. The only way to search is to try.
- **Fast to verify.** Once a miner finds a valid nonce, anyone on the network can check it with a single hash operation. The asymmetry — hard to find, easy to check — is what makes this useful.
- **Memoryless.** The probability of the next hash being valid doesn't depend on how many you've tried so far. (This will matter in the variance discussion below.)

If `T` is half the space of 256-bit integers, roughly 50% of nonces work, and you expect to find one after about 2 tries. If `T` is `2^(256-d)` for some `d`, the probability of any given nonce being valid is `2^-d`, and you expect to try `2^d` nonces. The quantity `d` — the number of leading zero bits — is the **difficulty**. Current Bitcoin difficulty corresponds to roughly `d ≈ 80` bits of work per block.

## Why the work is "wasted" — on purpose

This is the most-misunderstood part of Proof of Work, even among technical people, so it's worth being explicit.

The puzzle's solution has no intrinsic value. It's a random number that happens to be below a threshold. Nobody needs the number. The number is not used for anything downstream.

The *solution* is worthless; the *cost* is the point.

The cost is what makes a 51% attack expensive. The reason Bitcoin is hard to rewrite is not that the hashes are cryptographically magical — any computer can compute SHA-256 — but that producing a valid chain that's longer than the current one requires out-producing the entire rest of the network on raw compute. To rewrite the last `N` blocks, you must privately redo all `N` blocks' work while everyone else is doing ordinary new work. This is an arms race you can only win by having more hashrate than everyone else combined.

Put another way: the security budget of a Proof of Work chain is *the dollar value of the electricity and hardware being spent per unit time to keep it running*. If that budget falls, the cost of attacking falls with it. If the budget rises, so does the cost of attack. This is a real economic quantity, and we can put numbers on it.

## Difficulty adjustment

If the network's total hashrate doubles, blocks start arriving twice as fast. That would be bad — fee markets, propagation assumptions, and orphan rates all depend on a roughly stable inter-block time. So the protocol adjusts the difficulty periodically to keep blocks arriving at a target rate.

Bitcoin's formula, simplified: every 2,016 blocks (about 2 weeks at 10 minutes per block), the network measures how long those 2,016 blocks actually took and retargets difficulty to hit 10 minutes per block going forward:

<div class="math">

`new_difficulty = old_difficulty × (target_time / actual_time)`

</div>

If the last 2,016 blocks took 1 week (because hashrate doubled), difficulty doubles. If they took 4 weeks (hashrate fell), difficulty halves. Bitcoin caps adjustments at 4× in either direction per period to prevent wild swings.

This feedback loop is what keeps the block time roughly stable across eight orders of magnitude of hashrate growth since 2009. It's also why Bitcoin is said to be "self-regulating": nobody chooses the difficulty. It's a consequence of how much hashrate the network carries and how long blocks have recently taken.

## The probability math: when will the next block arrive?

Because each hash attempt is an independent Bernoulli trial with a fixed tiny probability `p` of success, and there are a huge number of attempts per second across the network, the time between blocks is well-modeled as an **exponential distribution**. If the network produces `λ` valid blocks per second on average, the time to the next block is:

<div class="math">

`T ~ Exponential(λ)`, with `E[T] = 1/λ` and `Var[T] = 1/λ²`.

</div>

For Bitcoin, `λ = 1/600` per second (one block every 10 minutes on average). So:

- Mean time to next block: **10 minutes** (by construction).
- Standard deviation of time to next block: **also 10 minutes**. This is a property of the exponential distribution — mean and standard deviation are equal.

The exponential has a long right tail. The probability that a given block takes more than `t` minutes is `e^(-t/10)`:

| t (minutes) | P(block takes ≥ t) |
|---|---|
| 10 | 36.8% |
| 20 | 13.5% |
| 30 | 5.0% |
| 60 | 0.25% |
| 120 | 0.0006% |

Half-hour gaps happen several times a week on Bitcoin. Hour-long gaps happen several times a year. This is normal and expected behavior, not a malfunction. It surprises many newcomers, who assume "10 minutes" means "10 minutes, give or take a minute."

The number of blocks produced in a fixed window is Poisson-distributed. In 60 minutes, you'd *expect* 6 blocks but could easily see 3 or 10. The variance is real, and it matters for confirmation math in Chapter 7.

## Orphans: what happens when two blocks arrive at once

If miner Alice and miner Bob both find a valid block at height `N+1` within a few seconds of each other, half the network will hear Alice's first and the other half Bob's. Both are valid. Neither can be "the" block until some later block gets built on top of one of them.

The moment a block at height `N+2` appears built on Alice's block, that chain is longer, and Bob's block becomes an **orphan** (also called a "stale" block). Bob's transactions go back to the mempool and are included in some later block. Bob loses the block reward; Alice keeps hers. No damage beyond that.

Orphan rate is roughly `propagation_time / block_time`. If blocks take about 1 second to propagate across the network and blocks are 10 minutes apart, you'd expect orphans in roughly 1/600 ≈ 0.17% of blocks, which matches Bitcoin's historical experience (≈0.1%–0.5% depending on conditions). Chains with much shorter block times (Ethereum pre-merge had 12-15 second blocks) see orphan rates closer to 5–10%, which is why their fork-choice and fee mechanics are more involved.

## What a 51% attack actually costs

This is the line you see everywhere: "It would cost billions to attack Bitcoin." Let's do the math, because the derivation is instructive and the number is genuinely large.

A 51% attack means acquiring more hashrate than the honest network combined. You don't need *exactly* 51% — more is better — but 51% is the threshold below which your success probability drops steeply. Assume for this estimate that you want to match current network hashrate, so you'd have 50% and win coin-flips. The honest-network's same hashrate is what you're matching.

**Hashrate to buy.** Bitcoin's total network hashrate as of early 2026 is approximately **700 EH/s** (exahashes per second, i.e. `7 × 10^20` hashes per second). Source: public estimates from blockchain.com / coinwarz / similar aggregators, which track this in real time. Like every number in this book, it's moving — check a current figure before making decisions based on it.

**Hardware efficiency.** The best-available Bitcoin ASIC miners in early 2026 achieve roughly **15–20 J/TH** (joules per terahash), with the Bitmain S21 family and similar units in that range. Source: manufacturer spec sheets. Call it 17 J/TH as a rough central estimate.

So powering 700 EH/s = 700,000,000 TH/s at 17 J/TH draws:

<div class="math">

`700,000,000 TH/s × 17 J/TH = 1.19 × 10^10 W ≈ 11.9 GW`

</div>

Matching this means running 11.9 GW of miners continuously. At an industrial electricity price of `$0.05/kWh`:

<div class="math">

`11.9 GW × 24 h × 365 d × $0.05/kWh ≈ $5.2 billion per year`

</div>

That's just the power. The hardware itself, at roughly **$20–$30 per TH of installed capacity** for current-generation units, implies:

<div class="math">

`700,000,000 TH × $25/TH ≈ $17.5 billion in ASICs`

</div>

And that's ignoring the fact that you can't actually buy that many ASICs at that price — trying to would distort the market and drive the price up significantly. Manufacturers produce on the order of tens of EH/s per month total; acquiring hundreds of EH/s of new hardware takes **years**, during which your target is also adding hashrate, meaning you have to keep buying to stay at 50%.

**Rent, don't buy?** A common objection: "you don't need to own the hardware, you can rent it." At short timescales, the public hashrate rental markets (e.g. NiceHash) carry only a few percent of Bitcoin's total hashrate — nowhere near enough for a 51% attack on Bitcoin, though that's been sufficient in the past for attacks on smaller PoW chains like Ethereum Classic and Bitcoin Gold, which have been attacked multiple times for exactly this reason. A Bitcoin 51% attack via rental markets is not currently available at any price.

**Why this is a realistic floor, not a ceiling.** The attacker needs the attack to last long enough to double-spend into something valuable (an exchange deposit, for instance) and get the reversal before the honest chain catches up. That takes several hours to days in practice, and during that time the attacker is burning electricity at the 11.9 GW rate *and* not earning legitimate block rewards (because they're mining a private fork that ultimately has to replace the public one). The realized cost is the electricity plus the opportunity cost of the block rewards plus the capital tied up in hardware. Actual analyses (Budish 2018, Moroz et al. 2020, and others) put the attack cost at many billions of dollars for a meaningful reorg window against Bitcoin, which matches the back-of-envelope above.

The more honest framing: **for Bitcoin specifically, at current hashrate, a 51% attack is economically irrational even for nation-state actors.** For smaller PoW chains, it absolutely is rational, has happened, and will happen again. Security scales with hashrate, which scales with block reward value, which scales with token price and transaction fees. Small PoW chains have small security budgets, which is precisely the structural weakness they inherit by choosing PoW as their consensus.

## Summary

Proof of Work is a Sybil-resistance mechanism that ties vote weight to burnt compute. The puzzle is SHA-256, the search is brute force by design, and the cost of the search is the cost of attack. Block times are exponentially distributed with high variance. Orphans are normal. Difficulty auto-adjusts to keep block times roughly constant as hashrate changes.

The famous number — "it would cost billions to attack" — is not a slogan; it's the actual arithmetic, and it's the thing PoW buys you. It also burns the electricity that went into producing it, which is a real cost that the ecosystem has to account for. Whether that cost is worth what it buys is the subject of Chapter 8.

Before we get there, we need to look at the main alternative: binding vote weight to capital instead of compute. That's Proof of Stake, and it's the next chapter.
