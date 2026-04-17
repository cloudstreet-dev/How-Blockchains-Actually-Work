# Finality: Probabilistic vs. Deterministic

"Wait for six confirmations." You've heard this; maybe you've said this. It's the folk wisdom for when a Bitcoin transaction is "safe." This chapter is about what that actually means, where the number comes from, and why other chains don't say it at all.

Finality is the property that, once a block is accepted into the chain, it will never be removed. That's the platonic version. In practice, finality comes in two flavors, and picking between them is one of the most load-bearing design decisions a blockchain makes.

## Probabilistic finality: Bitcoin's flavor

In Proof of Work, there is no moment at which a block is *definitely* final. There's only a moment at which it becomes *so unlikely to be reverted* that, for practical purposes, we treat it as final. The "six confirmations" rule is a probability calculation, and here's the calculation.

Suppose an attacker controls fraction `q` of the network's hashrate, and honest miners control the remaining `p = 1 - q`. The attacker's goal: you accept a payment (say, an exchange deposit) after `k` confirmations, and *before* you have a chance to credit or withdraw the funds, the attacker releases a longer private chain that didn't include their payment. If their chain becomes the longest, your payment is reversed.

For the attacker to succeed, they must privately produce blocks faster than the honest network, starting from some number of blocks behind. This is a race between two Poisson processes. The probability that the attacker ever catches up from being `k` blocks behind, given that they have hashrate fraction `q < 0.5`, is worked out in Satoshi's original paper (Section 11) and has the following closed form:

<div class="math">

`P(catchup | k blocks behind) ≈ (q/p)^k` (for `q < p`)

</div>

That's after the first block in the honest chain. Folding in the probability of how far behind they started (which itself depends on how many blocks the honest chain produced in the time the attacker was trying), the more complete formula — Satoshi's equation — is:

<div class="math">

`P(success) = 1 - Σ_{j=0}^{k} (λ^j · e^-λ / j!) · (1 - (q/p)^(k-j))` where `λ = k · (q/p)`.

</div>

You don't need to memorize this. You need to know the *shape* of the answer, which is: attack success probability falls off roughly geometrically with `k`, but how steeply depends on the attacker's hashrate share `q`. Satoshi's paper includes a table; here's a more thorough version, which we computed directly from the formula:

| Attacker's share `q` | Blocks needed for `P(success) < 0.1%` |
|---|---|
| 10% | 6 |
| 15% | 8 |
| 20% | 11 |
| 25% | 15 |
| 30% | 24 |
| 35% | 41 |
| 40% | 89 |
| 45% | 340 |

The "six confirmations" folk wisdom is approximately the right answer *if the attacker has at most about 10% of the hashrate*. For any serious attacker, six confirmations is not enough. Large exchanges know this — they use longer confirmation delays for large deposits, precisely because the tail risk grows fast as `q` rises. At roughly `q = 0.5`, no finite number of confirmations gives you any meaningful probability bound, because the attacker's expected chain-growth rate equals the honest network's. This is the "51% attack" threshold.

Note what this table is *not* saying. It's not saying an attack will happen — for Bitcoin, as we saw in Chapter 5, the cost makes it economically irrational. It's saying that if an attack happened, here's how long you'd have to wait to be safe. The cost of the attack is a separate line of defense. Probabilistic finality without the cost-of-attack being enormous is not much protection.

### A practical illustration

A $100 payment on Bitcoin with 1 confirmation: if an attacker has even 10% hashrate, there's a ~20% chance they can revert it. That's fine — nobody would spend millions of dollars of hashrate to steal $100.

A $100 million payment on Bitcoin with 6 confirmations: an attacker with 35% hashrate can revert this with ~25% probability per attempt (approximately). That attacker is losing, in expectation, many millions per hour of attack time in block rewards they could have legitimately earned. Is it profitable? It depends on the payment size relative to the attack cost. For a large enough payment, it could be. This is why exchanges require **many** more confirmations for large deposits — some use 100+ for the largest ones.

Probabilistic finality is usable. It is also, unambiguously, a probability — not a guarantee.

## Deterministic finality: BFT's flavor

Tendermint, HotStuff, Algorand, and their descendants give you the other thing: once a block is committed, it is **final, full stop**. Reverting it requires an attacker to control at least 1/3 of staked tokens *and* be willing to get them all slashed. There is no probability calculation, no number-of-confirmations rule, no "wait longer for bigger transactions." One commit, done.

The math underlying this is standard Byzantine-fault-tolerant agreement theory, going back to PBFT (Castro and Liskov, 1999) and earlier. The key property: if fewer than 1/3 of validators (by stake) are Byzantine, then:

- **Safety**: no two honest validators will ever commit conflicting blocks.
- **Liveness**: the network will continue to produce new blocks.

Both properties hold simultaneously under the 1/3 fault threshold. Above that threshold, safety and liveness can split: an adversary with >1/3 stake can either prevent progress (liveness failure) or, with >2/3, produce conflicting finalized blocks (safety failure).

The subtle point is that you can't have both safety and liveness under asynchrony — this is the FLP impossibility result (Fischer, Lynch, Paterson, 1985) — so BFT protocols make an assumption about the network (partial synchrony, typically) and provide guarantees under that assumption. In practice, the network is almost always synchronous enough for this to work, but when it isn't, you get chain halts. The Cosmos Hub and other Tendermint chains have occasionally halted for hours when validator networks became partitioned. The chain recovered cleanly; transactions just had to wait.

This is the big headline difference between BFT finality and Nakamoto-style finality:

- **BFT**: prioritizes safety. If it can't be sure no fork will exist, it stops. You may have to wait (liveness failure) but you will never see a reverted transaction (safety failure).
- **Nakamoto (Bitcoin, pre-merge Ethereum)**: prioritizes liveness. The chain always progresses. But a block you thought was final could, in principle, be reverted if enough hashrate attacks.

Neither is universally correct. Both are reasonable answers to different priorities.

## What about Ethereum's hybrid?

Ethereum after the Merge is a hybrid, as mentioned in Chapter 6. At the head of the chain, it uses LMD-GHOST, which is Nakamoto-flavored — blocks aren't truly final until attestation weight has piled up behind them. About every 6.4 minutes, Casper FFG finalizes a checkpoint, which gives you deterministic finality for all blocks up to that checkpoint.

This means Ethereum has **two** finality concepts:

- "Probabilistic" finality for the head, reached after a block gets enough attestations. A reorg of 1–2 blocks is possible in edge cases; anything deeper is extremely rare.
- "Economic" finality for finalized checkpoints. Once the checkpoint is finalized, reverting it costs ~1/3 of the stake (~$34B at current prices, as we computed in Chapter 6).

For user-facing applications, this hybrid means you pick your safety level based on the transaction value. For a small DEX swap, waiting a few blocks is fine. For a large institutional transfer, wait for the next checkpoint finalization (the `finalized` block tag in Ethereum's JSON-RPC) and you have BFT-grade guarantees.

## The tradeoff, summarized

| Property | Nakamoto (PoW) | BFT (PoS) | Hybrid (Ethereum) |
|---|---|---|---|
| Finality type | Probabilistic | Deterministic | Both (head + checkpoints) |
| Reorg possible? | Yes, always, geometrically unlikely | No, above fault threshold | Yes at head, no at checkpoint |
| Halts under partitions? | No — keeps growing | Yes — halts for safety | Yes — finality stalls |
| Time to "safe enough" | Minutes to hours | Seconds | Seconds to minutes |
| Requires known validator set? | No | Yes (bonded set) | Yes (bonded set) |

You can see why different applications, and different communities, have landed on different answers.

- A payment rail that prioritizes never going down but can tolerate minutes of settlement latency? Nakamoto.
- A financial application where settlement certainty is worth occasional outages? BFT.
- A smart-contract platform that wants cheap quick confirmation plus a way to reach economic finality for things that matter? Hybrid.

There's no winner in this comparison. There are design points on a curve, and different applications live at different points on the curve.

## Practical advice

If you are building on top of a blockchain:

- Know which finality your chain offers. Read the docs. If they say "1-second finality" without explaining what that means, push harder.
- Pick a confirmation rule appropriate to your transaction value. A dollar of exposure deserves a block; a million dollars of exposure deserves a checkpoint finalization.
- Never assume "finalized" means "can't be reverted by any means." In PoS, a sufficiently determined attacker spending enough money can always defeat any threshold; in PoW, the bound is probabilistic. In both cases, the real defense is that the cost is high, and you should know roughly what that cost is.

The last point, the cost, is the quantity the next chapter is about. We've now met both mainstream consensus families. Time to hold them up next to each other, honestly.
