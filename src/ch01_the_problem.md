# The Problem: Distributed Trust Without a Trusted Third Party

Before we can talk about how blockchains work, we need to be unreasonably clear about what problem they solve. Most of the confusion in the public conversation — including most of the skepticism and most of the evangelism — comes from being fuzzy on this point. If you get the problem right, the rest of the book is a long exercise in "well, given that, what else could you even do?"

The problem is this:

> **Agree on a shared, ordered history of events among a group of participants who don't trust each other, don't know each other, can join or leave at any time, and cannot all be assumed honest — without appointing anyone to be in charge.**

Every word in that sentence is load-bearing. Let's earn each one.

## Why this isn't the same problem as "databases"

If we had a trusted third party — a bank, a government registry, a well-run company — this problem would not exist. We'd hand them the history, they'd maintain it, and everyone would query them. This is how the world worked for most of computing history, and it works *extremely well*. A well-run SQL database, a Paxos-replicated key-value store, a cloud provider's transaction log — these are all solutions to the "shared ordered history" problem, and they're fast, efficient, and battle-tested.

The honest answer to "how do I keep a ledger of who owns what" has been, for most of the last century, *use a database run by someone trustworthy*. And if you can identify a trustworthy party who is going to stay trustworthy for the lifetime of your data, you should absolutely do that. It will be thousands of times more efficient than anything this book describes.

The problem blockchains attack is specifically the problem you're stuck with when **no such trusted party exists, or you don't want one, or you can't agree on one.**

That's a narrower problem than the marketing suggests. It is also a real problem, and before 2008 there was no clean computer-science answer to it.

## Double-spending: the clarifying example

The canonical version of this problem is *double-spending*, and it's the problem Bitcoin's original paper frames itself around. It's worth walking through slowly, because it explains why cryptography alone is not enough.

Suppose Alice has a digital coin, represented as some bytes signed by her. She wants to send it to Bob. She constructs a message:

> `Transfer(coin_42, from=Alice, to=Bob)` — signed by Alice.

Bob checks Alice's signature. The signature is valid. Bob concludes Alice has authorized this transfer. Great.

Except — what stops Alice from constructing a second message, five seconds later:

> `Transfer(coin_42, from=Alice, to=Carol)` — signed by Alice.

...and sending it to Carol? Carol also verifies the signature. It's also valid. Both Bob and Carol now believe they own `coin_42`.

Signatures prove *authorization*. They do not prove *uniqueness*. There is nothing about the second message that is cryptographically detectable as a double-spend, because it's a perfectly valid signed transfer. Bob and Carol, operating in isolation, have no way to know the other exists. Only a shared view of history — *has coin_42 already moved?* — can resolve the conflict.

So the double-spending problem reduces to: **can a group of participants agree on an ordered log, such that once a transaction is in the log, no later conflicting transaction can be accepted?** If yes, Bob checks the log, sees Alice's transfer to him, and can refuse any later transfer of the same coin. The log is what prevents the attack.

With a trusted third party, the log lives on their server. Easy. Without one, we're stuck. This is the hard problem, and it's the *only* problem we're solving. Everything else that blockchains are supposedly "good for" is a variation on this.

## Byzantine Generals: why even honest disagreement is hard

There's a classical formulation of why distributed agreement is difficult, and it predates blockchains by nearly three decades. It's called the Byzantine Generals Problem, from a 1982 paper by Lamport, Shostak, and Pease. It goes like this.

Several generals of the Byzantine army surround an enemy city. They must agree on a common plan — *attack* or *retreat*. If they all attack together, they win. If they all retreat, they survive. If some attack while others retreat, the attackers are slaughtered.

The generals can only communicate by messenger. Some of the generals may be traitors. A traitor will say anything to sow chaos: they might tell General A to attack while telling General B to retreat. Messages from loyal generals are truthful; messages from traitors are arbitrary lies.

The question: can the loyal generals always reach agreement, given that traitors might lie to them in any pattern?

The surprising answer is that if more than one-third of the generals are traitors, agreement is impossible in the general case. With `n` generals and `f` traitors, you need `n ≥ 3f + 1` loyal generals (really, `n ≥ 3f + 1` total, with `f` traitors) to reach agreement. This is a fundamental limit, not a weakness of any particular algorithm. It shows up in every consensus protocol that followed, under different names — "Byzantine fault tolerance" is the general label for tolerating these arbitrary-failure adversaries.

The point for us is that *even with honest participants, reaching agreement in the presence of some number of dishonest ones is nontrivial* — and the fraction of dishonest participants you can tolerate is bounded. There is no clever protocol that can tolerate 51% traitors. Not because nobody's invented it, but because it's been proven impossible.

This bound, `3f + 1`, will reappear in Chapter 6 when we look at modern BFT-style Proof of Stake. It's the reason those protocols talk about "two-thirds honest" and not "half honest." Bitcoin's Proof of Work, as we'll see in Chapter 5, gets a different bound (it only needs a majority of *hashpower*) at the cost of only giving probabilistic guarantees. Different tradeoff points on the same underlying problem.

## The missing ingredient: cost

Here's the move that changes everything, and it's the bit that was genuinely new in 2008.

Classical Byzantine agreement protocols assume a fixed, known set of participants. You have `n` generals, and you can count them. The protocols tolerate `f` faulty ones out of a known `n`.

But in an open system — one anyone can join — *you cannot count the participants*. Anyone can spawn a thousand identities and show up with a thousand votes. This is the Sybil attack, and it's the subject of the next chapter. It breaks every voting-based consensus algorithm in the classical literature when you try to run it on the open internet.

The problem we're actually trying to solve is therefore:

> Agreement on an ordered log, among an **unknown, open set** of participants, some fraction of whom are Byzantine-arbitrary-adversarial, **without appointing an authority**.

Every earlier attempt at digital cash — DigiCash, e-gold, Hashcash, b-money, bit gold — gave up one of those requirements. They either trusted a party, restricted the participant set, or accepted that the problem wasn't fully solved. Bitcoin's contribution, which we'll derive piece by piece over the next several chapters, was finding a combination of cryptographic and economic tricks that *does* close the last gap.

It closes it at a cost — real, measurable, and sometimes enormous. The book from here on is largely about what that cost is and when it's worth paying.

## What you should take away from this chapter

1. The problem is **open-membership Byzantine agreement on an ordered log**. That's it. If that problem doesn't apply to your situation, blockchains are the wrong tool.
2. Signatures alone do not solve this problem; double-spending is the one-line proof.
3. Classical Byzantine agreement gives us the `3f + 1` bound, but only for known participant sets.
4. Sybil attacks break naive voting in open systems, and dealing with them is the central innovation of what follows.

If you came in expecting "blockchains are a database," you now have the precise sense in which that's misleading: they're databases for the one specific case where you cannot rely on anyone to run the database. For every other case, use a database.
