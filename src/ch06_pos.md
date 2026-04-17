# Proof of Stake: How It Actually Works

Proof of Work uses burnt electricity as the scarce resource. Proof of Stake uses locked capital. That swap is the whole idea — and as with most one-line summaries of complicated systems, it's also slightly misleading. The details are where this chapter lives.

## The core idea, in one paragraph

To participate in consensus, you deposit some of the system's native token into a protocol-controlled contract. That deposit is your **stake**. The protocol lets participants with larger stakes produce and vote on blocks more often, proportional to stake size. Behave honestly, and you earn rewards — more stake over time. Behave demonstrably dishonestly (double-sign a block, vote for two conflicting forks), and the protocol **slashes** your stake: a portion of it is burned or redistributed. Sybil attacks don't help, because splitting a stake across more identities doesn't give you more stake, and buying more stake means actually buying more tokens on the open market, which is necessarily expensive.

That's it at the idea level. Now for what goes wrong when you try to build it.

## The "nothing at stake" problem

Proof of Work has a pleasant property: mining on a fork is *costly*. Every hash you throw at the wrong fork is a hash you don't throw at the right one. Miners are therefore economically pushed to pick one fork and commit to it.

Naive Proof of Stake has the opposite property. Signing a vote on a fork is effectively free — it's a few bytes and a signature. A rational validator might as well sign *every* fork they see, because if one of them ends up winning, they've voted for the winner; if all of them lose, they've lost nothing. This is the "nothing at stake" problem: naive PoS has no mechanism to discourage double-voting, so forks multiply and consensus never converges.

The fix is the one you'd reach for: **slashing.** The protocol builds in rules like "if you sign two conflicting blocks at the same height, anyone who observes both can submit a proof and you lose a large fraction of your stake." Suddenly double-voting is extremely costly. Validators are economically pushed to pick one fork, same as miners.

Slashing is the most important mechanism in modern Proof of Stake. Everything else — validator rotation, reward curves, block production — is filling in around this load-bearing piece. The historical PoS designs that failed to take slashing seriously (early Peercoin, for instance) had to add ad-hoc anti-fork mechanisms like checkpoints signed by the dev team, which was an obvious retreat from the security model.

## Long-range attacks and weak subjectivity

Here's a subtler problem, and the one that made cryptographers skeptical of PoS for a long time.

Suppose you're a new node joining an established chain. You download all the blocks from genesis and want to verify you're on the right chain. In Proof of Work, this is easy: you sum up the total work of the chain you're verifying, and if it's the heaviest, it's the right one. Work is *real in the world* — you can't fake it, you can't retroactively produce it, and you don't need anyone's help to check it.

In Proof of Stake, the analogous check doesn't exist. An attacker who controlled a majority of the stake *years ago* — at some point in the distant past, before they sold their tokens — could in principle use those old keys to forge an alternate history from that point forward, producing a chain that looks perfectly valid to a new node with no prior knowledge. The validators who signed those historical blocks don't have any stake at risk anymore, because they cashed out. Slashing doesn't help, because there's nothing to slash. This is the **long-range attack**.

The fix is called **weak subjectivity**. New nodes (or nodes that have been offline for a long time) need one piece of out-of-band information to safely sync: a recent *checkpoint* — a block hash known to be on the real chain, obtained from a trusted source (your friend, a block explorer, the protocol's official docs, etc.). Once you have that anchor, you can verify everything from it onwards using ordinary stake-weighted voting, because the stake *today* is at real risk of slashing.

This is the honest tradeoff that skeptics used to point to: Proof of Work is "objectively verifiable from genesis"; Proof of Stake requires a weakly-subjective starting point. Whether that's a meaningful security difference in practice is a long debate. The practical consensus in the field today is: **for an actively used chain with a lot of stake online, weak subjectivity is a one-time sync-time concern that doesn't affect normal operation**, and it's acceptable. For a chain that's dormant or has most of its validators offline, it becomes a more serious concern.

## Validator selection

A PoS protocol has to answer: at any given moment, who gets to propose the next block?

"Everyone who has stake, proportional to their stake" is too vague — it doesn't pick a single proposer, and you can't have everyone proposing at once without creating forks. The protocol needs a way to pseudo-randomly pick *one* validator per slot, in a way that:

1. Can't be manipulated by the validators themselves (you can't game your way into being selected).
2. Is verifiable by everyone (so the rest of the network can check that the proposer was, in fact, selected for this slot).
3. Is roughly proportional to stake, over time.

Modern designs use **verifiable random functions (VRFs)** or **commit-reveal schemes with randomness beacons** for this. The inputs to the randomness are things the validator can't influence — typically the hash of the previous block combined with some protocol-level entropy — and the output is a deterministic-but-unpredictable assignment of slots to validators.

In Ethereum specifically, the randomness source is called **RANDAO**, and it's accumulated over many blocks by having each proposer contribute a fresh random value. Manipulating it requires controlling many consecutive slots, which requires a large fraction of the stake. The design is not perfect — there's a known manipulation in which the proposer of the last slot of an epoch can choose whether to reveal their contribution, giving them a one-bit influence over the next epoch's randomness — but in practice the cost of exploiting it is much higher than the benefit except in edge cases.

## Two families of Proof of Stake

In practice, production PoS systems divide into two camps, which have quite different properties.

### Chain-style PoS (exemplified by Ethereum)

Ethereum's consensus since the September 2022 Merge is **Gasper**, a combination of two sub-protocols:

- **LMD-GHOST** (Latest Message Driven — Greediest Heaviest Observed SubTree) handles ongoing block production. Validators attest to the block they think should be the head of the chain; the fork choice follows the subtree with the most attestation weight. This gives Ethereum "probabilistic" finality block-by-block, similar in spirit to Bitcoin's longest-chain rule but weighted by attestations instead of work.

- **Casper FFG** (Friendly Finality Gadget) handles *economic* finality. Every ~6.4 minutes (an epoch), validators vote on checkpoint blocks. Once two-thirds of stake has voted for a checkpoint in two successive epochs, that checkpoint is **finalized** — reverting it would require slashing a third of the stake, which is an enormous amount of money (tens of billions of dollars at typical ETH prices).

So Ethereum has *both* a probabilistic head (which is reorg-able over short timescales) and a true finality layer (which is not, absent an economically catastrophic attack). This is a hybrid design, and it's quite different from either pure Nakamoto consensus or pure BFT.

### BFT-style PoS (exemplified by Tendermint / CometBFT, used by Cosmos)

The other family does full Byzantine-fault-tolerant agreement every single block. Tendermint (now called CometBFT in the Cosmos ecosystem) runs a three-phase vote — **pre-vote**, **pre-commit**, **commit** — among a known set of validators (typically ~100–200). A block is final the instant two-thirds of stake-weighted validators commit to it. There are no orphans, no reorgs, no "wait for six confirmations" — the block either gets 2/3 commit and is final, or it doesn't, and the network moves to the next round with a different proposer.

Tendermint gives you **deterministic finality within about 1–7 seconds** at the cost of requiring a known validator set (so it's not as open as Ethereum — Cosmos chains typically use a bonded, permissionless validator set capped at a few hundred slots) and being somewhat more fragile under high latency: if more than 1/3 of stake is offline or unreachable, the chain halts (**liveness failure**) rather than producing wrong blocks (**safety failure**). This is called "halting over advancing in uncertainty," and for many applications — especially financial ones — it's the correct tradeoff.

Other BFT-style PoS designs: HotStuff (which Aptos, Sui, and the old Diem all derive from), Algorand's agreement protocol, and Solana's Tower BFT (with its own idiosyncrasies — more in Chapter 11). The family is broad but they all share the core property of deterministic per-block finality with a known validator set.

## The staking economics

The security budget of a PoS chain is the total value of staked tokens, weighted by the slashing fractions. Roughly: an attacker who wants to cause a safety failure (two finalized conflicting chains) must control one-third of the stake and be willing to lose it.

As a concrete example: Ethereum in early 2026 has roughly 34 million ETH staked out of ~120 million total supply — about 28% of supply. At an ETH price of roughly $3,000 (highly volatile, so use the current price for real decisions), that's **~$102 billion of stake at risk**. To cause a safety failure, an attacker needs to acquire and then forfeit one-third of that — roughly **$34 billion** — which is, as with Bitcoin, more than any rational attacker would spend. (Source for staking figures: beaconcha.in and public Ethereum dashboards, as of early 2026; ETH price is whatever it happens to be when you read this.)

Note the parallel with the PoW attack cost in Chapter 5: both mainstream chains have security budgets in the tens of billions of dollars, and both are economically irrational to attack at scale. The *shape* of the cost is different — ongoing electricity for PoW, one-time capital at risk for PoS — which matters for the environmental footprint, centralization pressures, and other concerns we take up in Chapter 8.

## Where PoS has rough edges

Proof of Stake is not a finished subject. Active concerns include:

- **Validator centralization through liquid staking.** Services like Lido (Ethereum), Jito (Solana), and others pool stake from thousands of users and run validators on their behalf. Lido alone has hovered around 28–30% of Ethereum stake — right at the edge of a concerning concentration for a protocol where 33% can halt finality. The protocol's rules don't discourage this; the social layer does. Whether the social layer is a reliable security mechanism is a real and open question.

- **MEV (Maximal Extractable Value).** Block proposers get to decide not just which transactions are included but in what order. In a world with DeFi, transaction ordering can be worth millions per day. This incentivizes all sorts of emergent behavior (off-protocol proposer auctions, specialized "builders" who construct high-MEV blocks for proposers, etc.) that Ethereum is still trying to figure out how to handle cleanly. Proposer-Builder Separation (PBS) is the current leading design.

- **Validator exit queues.** Most PoS designs rate-limit how fast validators can unstake, to prevent coordinated mass exits. This limits the chain's agility in response to attacks and adds friction to the staking UX, but it's essential: without it, an attacker could attack and then exit their stake before being slashed.

None of these are deal-breakers, but they're the frontier, not a solved problem. When a marketer tells you PoS is "strictly better than PoW," ask them which version of PoS, how it handles liquid staking concentration, and what its MEV story is. If they don't have an answer, they're selling.

## Summary

Proof of Stake ties vote weight to locked capital instead of burnt compute. Slashing makes double-voting expensive; validator selection uses verifiable randomness; long-range attacks are mitigated with weak-subjectivity checkpoints. Two main families — Nakamoto-adjacent hybrid (Ethereum's Gasper) and pure BFT (Tendermint and successors) — occupy different points on the finality/liveness curve.

The core tradeoffs against Proof of Work — energy cost vs capital lockup, probabilistic security vs economic security, bootstrapping difficulty vs weak-subjectivity — are the subject of Chapter 8. First, though, we need to be precise about what "final" means on either side, because the word hides more than it reveals. That's Chapter 7.
