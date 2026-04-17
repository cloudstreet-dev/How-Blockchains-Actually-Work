# Scaling: Rollups, L2s, and What They Actually Do

The last chapter ended with an observation: a smart-contract chain's throughput is bounded by what a single validator can do, because every validator does the same work. For Ethereum that's roughly 15–25 transactions per second, give or take, depending on transaction complexity.

Raising this limit on the base chain is a political and technical minefield. Every time you make a block larger or faster, you raise the hardware cost of running a validator, which pushes some of them offline, which *reduces* decentralization. Bitcoin had a multi-year civil war over whether to raise its block size from 1MB, and the answer was, effectively, "no, we'll scale on a second layer." Ethereum has been less categorical, raising its block gas limit periodically, but the same structural argument applies: you can't make the base chain much faster without changing what "running a node" means.

The modern answer to scaling, on both Bitcoin-style and Ethereum-style chains, is **layer 2**: do the actual work somewhere else, and use the base chain as a settlement and dispute-resolution layer. The details of how this works are the subject of this chapter.

## The big idea

A layer 2 (L2) is a separate system that:

1. Processes transactions off the base chain (fast, cheap).
2. Periodically posts a *commitment* to the base chain (slow, expensive, but small: one transaction covers many L2 transactions).
3. Provides a way to *verify* that the commitment is correct, so the base chain can enforce good behavior.

The commitment is usually a hash of the new L2 state root (or of a batch of L2 transactions) published as the payload of a base-chain transaction. The verification is the interesting part, and it's the dimension on which the two main L2 families diverge.

## Optimistic rollups

Assume the L2 operator is honest; if they're not, let anyone prove it on-chain and slash them. That's the "optimistic" in optimistic rollup.

The mechanics:

1. The operator (called a **sequencer**) collects L2 transactions, executes them, and posts a new state root to L1. The root represents the L2 state after applying the batch.
2. The root is not accepted as final immediately. There's a **challenge window**, typically 7 days.
3. During the window, anyone with the means to do so can download the batch, re-execute it, and submit a **fraud proof** if they find the state root doesn't match the correct execution. The fraud proof is a cryptographic argument, submitted as an L1 transaction, that lets the L1 contract determine whether the sequencer cheated. If so, the sequencer loses a bond; the bad state root is rolled back; the correct one is substituted.
4. After the window expires with no successful challenge, the root is final.

Optimism and Arbitrum are the two leading optimistic rollup implementations; both have billions of dollars of value settled on them.

The 7-day delay is the cost: if you withdraw funds from the L2 to L1, you have to wait a week (or use a third-party bridge that fronts the money, charging a fee, taking the delay risk themselves). For DEX trading and normal activity on the L2 itself, there's no wait — the 7 days only matters for L2→L1 exits.

The security assumption: **at least one honest actor with fast enough access must be willing and able to produce a fraud proof.** If the operator is cheating and *no one* challenges in 7 days, the fraud goes through. This is a "1-of-N honesty" assumption, which is much weaker than any consensus protocol's majority assumption, but it's not "trustless" in the strongest sense.

## ZK rollups

Don't assume; prove. A zero-knowledge rollup submits, along with each batch, a cryptographic proof that the new state root is the correct result of applying the batch to the old state root. The proof is checked by an L1 contract at submission time; if it verifies, the root is accepted as final immediately.

The proof is a **zk-SNARK** or **zk-STARK** — different proof systems with different tradeoffs. Both have the property that verifying the proof is cheap and fast (tens of thousands of gas on L1, regardless of batch size), while generating the proof is expensive (seconds of computation off-chain, sometimes on GPU clusters).

Leading ZK-rollup implementations include zkSync Era, Starknet, Scroll, Linea, and Polygon zkEVM. They differ in exactly which EVM opcodes they support, how efficiently they do it, and how their proof systems are structured. Starknet uses its own non-EVM language (Cairo) specifically because proving arbitrary EVM is harder than proving Cairo, and the tradeoff has been visible in the pace of rollout.

The name "zero-knowledge" is historical and somewhat misleading. What matters for rollups is the *succinctness* of the proof — the ability to verify "I did this computation correctly" in a small amount of work. The "zero-knowledge" part (the prover learns the result without revealing the inputs) isn't actually used in rollups; the inputs are public. The field calls them "validity rollups" in more careful writing, but the name ZK rollup has stuck.

The tradeoff vs. optimistic: ZK rollups have **instant L1-finality** (no 7-day challenge window, once the proof is posted), at the cost of proving being expensive enough that batch submissions are less frequent and there's meaningful proving latency. Both tradeoffs are actively narrowing as proof systems improve, and most observers expect ZK rollups to dominate over the 2025-2030 timeframe. As of early 2026, optimistic rollups still carry more TVL (total value locked) than ZK rollups, but that gap has been closing.

## The trust assumption spectrum

Not everything marketed as "L2" is actually a rollup. There's a gradient:

- **Rollup**: all L2 transaction data is posted to L1 (as calldata or blob data, see below), so anyone can reconstruct state independently. This is the strongest variant.
- **Validium**: state transitions are proven on L1, but transaction data is held off-chain by the operator. Cheaper, but if the operator goes offline and refuses to release the data, users can't reconstruct their balances to exit. StarkEx (the underlying engine for DYDX, earlier ImmutableX) uses this.
- **Optimium**: optimistic rollup analogue of validium. Same fraud-proof model, but data is off-chain. Same availability risk.
- **"Layer 2" in the marketing sense**: anything that claims to extend a base chain's reach. Many sidechains (Polygon PoS, Binance Smart Chain) are their own independent chains with their own validators, connected to Ethereum only by bridges. These are *not* rollups and do *not* inherit Ethereum's security; they inherit the security of their own validator sets, which are usually much smaller.

If someone tells you their L2 is "just as secure as Ethereum," check: **are all the transaction data posted to L1, and is the state-transition validity guaranteed by L1?** If not, then no, it isn't. That's fine — there are sensible applications for validiums and sidechains — but it's not what a rollup is.

## Data availability: the actual bottleneck

Here's the punchline that surprises people. Computation is *not* the scaling bottleneck anymore. For ZK rollups especially, proving an enormous batch of transactions costs only one L1 verification, and the L1 verification is cheap.

The bottleneck is **data availability**: the cost of publishing the raw transaction data on L1 so anyone can reconstruct the L2 state. Even ZK rollups publish this data, because *without* it, a byzantine operator could post a valid proof for a state you can't reconstruct, and you'd be unable to exit. You'd know your state was "legal" but not what it was.

For a long time, L1 calldata was expensive (16 gas per non-zero byte, 4 per zero byte). Ethereum's EIP-4844 (March 2024), known as "proto-danksharding" or "blobs," introduced a separate data-availability lane: each block can carry up to ~6 "blobs" of ~128 KB each, with a separate fee market that's much cheaper than calldata. Blobs are pruned after about 18 days, which is long enough for any honest party to download and re-publish them if needed, but short enough that nodes don't have to store them forever.

This has been the single biggest scaling win for Ethereum L2s. After EIP-4844, rollup transaction fees dropped by roughly 10× on average. Blob capacity will continue to expand in future upgrades ("danksharding" is the endgame target, with on the order of 100+ blobs per block).

For applications with even higher data throughput needs, dedicated data availability layers (Celestia, EigenDA, NEAR DA) have emerged. They offer cheaper DA at the cost of weaker security assumptions than Ethereum's main DA layer. The tradeoff in choosing between them is, again, not "which is better" but "which security/cost point do you need."

## What an L2 transaction looks like end-to-end

Walking through a typical optimistic-rollup trade from a user's view:

1. User opens a wallet connected to the L2 (e.g. Arbitrum One). The wallet shows their L2 balance.
2. User submits a swap. The transaction goes to the L2 sequencer, which orders and executes it essentially instantly (block time ~250 ms) and returns a receipt.
3. The sequencer batches this transaction with many others and posts a rollup-batch transaction to Ethereum L1, containing (post-EIP-4844) the batch data as blobs and the new state root as calldata.
4. The batch is now on L1. For seven days, it could be challenged. During these seven days, the user's transaction is "L2-final" (the sequencer has confirmed it, the data is public) but not yet "L1-final" (the state root could still be rolled back by a successful fraud proof).
5. No challenge is filed. After seven days, the state root is final and irreversible. If the user wants to bridge assets back to L1, they can now do so directly from the rollup contract.

From the user's perspective, this feels like using any other chain — transactions confirm in a fraction of a second. The 7-day L1-finality is invisible except for the specific case of withdrawing *to* L1, which most users don't do often.

## What's actually hard about L2s

The rhetoric around L2s sometimes makes it sound like the scaling problem is solved. It isn't, quite. Current open questions:

- **Sequencer centralization.** Most rollups today have one sequencer, run by the rollup team. That sequencer can censor transactions (though users can usually force-include via L1), reorder them for MEV, or go offline and halt the rollup. Decentralizing sequencers is an active design problem — shared sequencers (Espresso, Astria) and round-robin models are both being tried.
- **Interoperability across L2s.** If Alice is on Optimism and Bob is on Arbitrum, paying Bob requires a bridge — and bridges remain the most-exploited part of the ecosystem. Fast cross-rollup interop is being worked on but isn't solved.
- **State growth.** Even with blob data pruned after 18 days, the *state* of each rollup grows indefinitely. Long-term this is a problem for everyone, including L1.
- **Prover centralization (ZK rollups).** Proving is computationally heavy and increasingly done by specialized operators. Decentralizing provers so anyone can generate proofs is an active problem.

None of these are showstoppers. All of them are the kind of thing that, if left unaddressed, subtly reintroduces centralization through the back door. The next few years of blockchain engineering are mostly about these problems.

## Summary

Base chains scale badly on purpose: raising throughput raises node hardware cost raises centralization. The modern answer is rollups, which do work off-chain and submit succinct summaries to the base chain. Optimistic rollups bet on fraud proofs and a 7-day challenge window; ZK rollups use validity proofs and get immediate finality at the cost of proving overhead. The real bottleneck in both is data availability, and blob-based DA (EIP-4844 on Ethereum) was the recent big unlock.

Not every "L2" is a rollup. Some are validiums; some are sidechains; some are rebranded payment channels. Know what you're looking at, and ask how much of the base chain's security is actually inherited.

Now that we understand both the consensus mechanisms and the scaling mechanisms, the next chapter is a quick tour of the actual production chains you'll encounter: who uses what, why, and what's marketing vs. what's meaningful engineering difference.
